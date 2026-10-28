# Google Drive API com Python

Este projeto demonstra como conectar uma aplicação Python à **Google
Drive API v3** para listar, consultar, enviar, baixar, renomear, mover,
excluir e compartilhar arquivos do Google Drive.

A integração é útil para automatizar tarefas, organizar documentos e
incorporar o Google Drive a scripts Python, aplicações Streamlit ou APIs
com FastAPI/Flask.

## 1. Como funciona

A aplicação Python envia solicitações à Google Drive API. Para acessar
arquivos privados, é necessário autenticar a aplicação e obter as
permissões adequadas.

Este guia utiliza **OAuth 2.0 para aplicativo de computador**. Na
primeira execução, o navegador será aberto para que o usuário entre na
conta Google e autorize o acesso. Depois, o token pode ser reutilizado.

> Uma credencial OAuth identifica a aplicação; ela não concede acesso
> automaticamente a todos os arquivos. O acesso efetivo depende da conta
> autenticada, dos escopos e das permissões do Google Drive.

## 2. Preparar o projeto no Google Cloud

1.  Acesse o [Google Cloud Console](https://console.cloud.google.com/).
2.  Crie ou selecione um projeto.
3.  Abra **APIs e serviços → Biblioteca** e ative a **Google Drive
    API**.
4.  Configure a **Google Auth Platform** (ou a tela de consentimento
    OAuth, dependendo da interface).
5.  Se o aplicativo estiver em modo de teste, abra **Público-alvo
    (Audience)** e adicione seu endereço Google em **Usuários de
    teste**.
6.  Em **Clientes** ou **Credenciais**, crie um ID de cliente OAuth do
    tipo **App para computador (Desktop app)**.
7.  Baixe o arquivo JSON e salve-o na pasta do projeto com o nome
    `credentials.json`.

Se aparecer a mensagem de que o aplicativo está em fase de testes e só
pode ser acessado por testadores aprovados, confira se a conta com a
qual está tentando entrar foi adicionada à lista de usuários de teste.

Para um projeto pessoal em desenvolvimento, normalmente não é necessário
publicar o aplicativo para todos os usuários. A verificação e as regras
aplicáveis dependem dos escopos solicitados e do tipo de distribuição.

Documentação oficial: [Quickstart da Google Drive API para
Python](https://developers.google.com/workspace/drive/api/quickstart/python).

## 3. Estrutura sugerida

``` text
google-drive-python/
├── .venv/
├── credentials.json   # Credencial OAuth baixada do Google Cloud
├── token.json         # Criado após a autorização inicial
├── drive.py           # Código Python
├── requirements.txt
├── .gitignore
└── README.md
```

**Não compartilhe nem publique `credentials.json` ou `token.json`.**
Eles contêm informações sensíveis. Se uma chave ou segredo for exposto,
revogue ou recrie a credencial afetada no Google Cloud.

## 4. Instalação

Requer Python 3.10 ou superior recomendado para este exemplo.

No Ubuntu/Linux:

``` bash
mkdir google-drive-python
cd google-drive-python

python3 -m venv .venv
source .venv/bin/activate

pip install --upgrade google-api-python-client google-auth google-auth-httplib2 google-auth-oauthlib
```

Crie um arquivo `requirements.txt` com:

``` text
google-api-python-client
google-auth
google-auth-httplib2
google-auth-oauthlib
```

Se o projeto já existe, basta ativar seu ambiente virtual e instalar as
dependências.

## 5. Conectar ao Google Drive

Crie `drive.py` com o código abaixo. O escopo `drive` permite gerenciar
arquivos aos quais a conta autenticada tem acesso. Para aplicações que
só precisam ler arquivos, considere usar um escopo mais restrito, como
`drive.readonly`.

``` python
import os.path

from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

SCOPES = ["https://www.googleapis.com/auth/drive"]


def conectar_drive():
    credenciais = None

    if os.path.exists("token.json"):
        credenciais = Credentials.from_authorized_user_file(
            "token.json", SCOPES
        )

    if not credenciais or not credenciais.valid:
        if (
            credenciais
            and credenciais.expired
            and credenciais.refresh_token
        ):
            credenciais.refresh(Request())
        else:
            fluxo = InstalledAppFlow.from_client_secrets_file(
                "credentials.json", SCOPES
            )
            credenciais = fluxo.run_local_server(port=0)

        with open("token.json", "w", encoding="utf-8") as token:
            token.write(credenciais.to_json())

    return build("drive", "v3", credentials=credenciais)


drive = conectar_drive()
print("Conexão com a Google Drive API estabelecida.")
```

Execute:

``` bash
python drive.py
```

Na primeira execução, uma janela do navegador será aberta para
autenticação e autorização. Depois da autorização, o programa cria
`token.json` para reutilizar as credenciais. Não é necessário repetir o
login a cada execução enquanto o token continuar válido e puder ser
renovado.

### Observações sobre o modo de teste

-   A conta Google usada no navegador deve estar cadastrada como usuária
    de teste quando o aplicativo estiver em modo de teste.
-   Em aplicações externas, tokens de atualização associados a
    aplicativos externos em modo de teste podem expirar após sete dias,
    dependendo das condições e dos escopos.
-   Se você mudar os escopos solicitados, pode ser necessário remover o
    `token.json` antigo e autorizar novamente.
-   Não apague o token sem necessidade: isso fará o fluxo de
    autenticação ser solicitado novamente.

## 6. Operações comuns

Os exemplos seguintes pressupõem que `drive = conectar_drive()` já foi
executado.

### Listar arquivos

``` python
resultado = drive.files().list(
    pageSize=20,
    orderBy="name",
    fields="files(id, name, mimeType, webViewLink)"
).execute()

for arquivo in resultado.get("files", []):
    print(arquivo["name"], arquivo["id"], arquivo["mimeType"])
```

A resposta contém metadados e IDs. Use o `id` para operar sobre um
arquivo específico. Para listar muitos arquivos, implemente paginação
com `nextPageToken`.

### Consultar metadados

``` python
arquivo = drive.files().get(
    fileId="ID_DO_ARQUIVO",
    fields="id, name, mimeType, parents, webViewLink"
).execute()

print(arquivo)
```

Substitua `ID_DO_ARQUIVO` pelo ID real.

### Criar uma pasta

``` python
pasta = drive.files().create(
    body={
        "name": "Projetos Python",
        "mimeType": "application/vnd.google-apps.folder"
    },
    fields="id, name"
).execute()

print(pasta)
```

Para criar uma subpasta, informe o ID da pasta pai em `parents`:

``` python
subpasta = drive.files().create(
    body={
        "name": "Relatorios",
        "mimeType": "application/vnd.google-apps.folder",
        "parents": [pasta["id"]]
    },
    fields="id, name"
).execute()
```

### Enviar um arquivo (upload)

``` python
from googleapiclient.http import MediaFileUpload

arquivo = drive.files().create(
    body={"name": "relatorio.csv"},
    media_body=MediaFileUpload(
        "relatorio.csv",
        mimetype="text/csv"
    ),
    fields="id, name, webViewLink"
).execute()

print(arquivo)
```

Para enviar o arquivo para uma pasta específica, adicione
`"parents": ["ID_DA_PASTA"]` ao `body`.

### Baixar um arquivo

``` python
from googleapiclient.http import MediaIoBaseDownload

file_id = "ID_DO_ARQUIVO"
request = drive.files().get_media(fileId=file_id)

with open("arquivo_baixado.pdf", "wb") as destino:
    downloader = MediaIoBaseDownload(destino, request)
    concluido = False

    while not concluido:
        status, concluido = downloader.next_chunk()
        if status:
            print(f"{int(status.progress() * 100)}%")
```

Esse método serve para arquivos binários comuns, como PDF, CSV e TXT.
Documentos nativos do Google Docs, Sheets e Slides normalmente precisam
ser exportados usando `files().export()` para um formato compatível.

### Renomear um arquivo

``` python
drive.files().update(
    fileId="ID_DO_ARQUIVO",
    body={"name": "relatorio_final.pdf"}
).execute()
```

### Mover um arquivo para outra pasta

``` python
file_id = "ID_DO_ARQUIVO"
nova_pasta_id = "ID_DA_NOVA_PASTA"

arquivo = drive.files().get(
    fileId=file_id,
    fields="parents"
).execute()

pais_atuais = ",".join(arquivo.get("parents", []))

drive.files().update(
    fileId=file_id,
    addParents=nova_pasta_id,
    removeParents=pais_atuais
).execute()
```

Esse exemplo é voltado a arquivos com uma pasta pai convencional. Drives
compartilhados podem exigir parâmetros adicionais, como
`supportsAllDrives=True`.

### Compartilhar um arquivo

``` python
drive.permissions().create(
    fileId="ID_DO_ARQUIVO",
    body={
        "type": "user",
        "role": "reader",
        "emailAddress": "pessoa@exemplo.com"
    },
    fields="id"
).execute()
```

Troque o endereço de exemplo pelo endereço real. A operação depende das
permissões do usuário e das políticas da organização.

### Excluir um arquivo

``` python
drive.files().delete(
    fileId="ID_DO_ARQUIVO"
).execute()
```

**Cuidado:** essa operação exclui permanentemente o arquivo. Confirme o
ID antes de executá-la. Se quiser apenas movê-lo para a lixeira,
atualize o campo `trashed` para `True` em vez de usar `delete()`.

## 7. Referência rápida

  Operação                  Método principal
  ------------------------- ----------------------------------------
  Listar arquivos           `files().list()`
  Consultar metadados       `files().get()`
  Criar pasta ou arquivo    `files().create()`
  Enviar arquivo            `files().create()` + `MediaFileUpload`
  Baixar arquivo            `files().get_media()`
  Renomear ou atualizar     `files().update()`
  Mover entre pastas        `files().update()`
  Excluir permanentemente   `files().delete()`
  Compartilhar              `permissions().create()`

Referência: [Google Drive API v3 --- documentação
oficial](https://developers.google.com/workspace/drive/api/reference/rest/v3).

## 8. Segurança

Adicione ao `.gitignore`:

``` gitignore
.venv/
__pycache__/
.env
credentials.json
token.json
```

Boas práticas:

-   Nunca publique credenciais ou tokens no GitHub.
-   Solicite apenas os escopos necessários.
-   Evite excluir ou compartilhar arquivos sem validar os IDs e as
    permissões.
-   Em aplicações hospedadas, guarde segredos em variáveis de ambiente
    ou em um gerenciador de segredos; não dependa de abrir um navegador
    interativo no servidor.
-   Se um segredo for exposto, revogue ou gere novamente a credencial
    afetada.

## 9. Problemas comuns

**"O aplicativo está em fase de testes"**

Adicione a conta Google usada no login à lista de usuários de teste na
seção **Público-alvo** da Google Auth Platform.

**"Access denied" ou arquivo não encontrado**

Confirme que está autenticado com a conta correta, que o ID está correto
e que a conta tem permissão para acessar o arquivo.

**"Insufficient Permission"**

O token pode não conter o escopo necessário. Revise `SCOPES`; se alterar
os escopos, refaça a autorização quando necessário.

**O token parou de funcionar**

Pode ter expirado, sido revogado ou estar sujeito às limitações do modo
de teste. Execute novamente o fluxo de autenticação e verifique a
configuração do aplicativo.

## 10. Próximos passos

Depois de validar a conexão, você pode encapsular as operações em
funções próprias, criar uma interface Streamlit para navegar pelas
pastas ou disponibilizar as operações por uma API FastAPI.

-   [Quickstart oficial para
    Python](https://developers.google.com/workspace/drive/api/quickstart/python)
-   [Referência da Google Drive API
    v3](https://developers.google.com/workspace/drive/api/reference/rest/v3)
-   [Guia de autorização e
    escopos](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
