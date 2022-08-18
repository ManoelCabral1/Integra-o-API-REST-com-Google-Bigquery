# Integração API REST com Google Bigquery :man_technologist:

APIs REST são uma das fontes de dados mais comuns no dia a dia de trabalho do analista de dados. São por meio delas que a maioria das plataformas sejam redes sociais, CRM, ecomerce, etc disponibilizam seus dados. Contudo uma dificuldade desse modelo de dados é que o formato padrão JSON, pode não ser tão intuitivo e fácil de interpretar como o formato tabular de planilhas ou tabelas em banco de dados.

Pensando em facilitar o acesso, o armazenamento  e interpretação do dados para um projeto onde se fazia necessário a coleta diária de informações em uma API. Projetei um pipeline integrando a API ao google bigquery (que uma plataforma na nuvem sem servidor para análise de dados).

## Requisitos para o projeto
1. A coleta será diária e incremental sendo adicionada a tabela somente os dados referentes a d-1, isto é, a data da coleta menos um dia.
2. A API tem endpoints (url de requisição) baseada em intervalos de data, logo deve haver um meio de persistir no banco somente os dados da data de interesse. 
3.  o processo deve ser automático  e resiliente a falhas.

## Solução
 A maneira mais conveniente  que encontrei para integrar a API ao bigquery  foi via cloud functions. O Google Cloud Functions constitui uma plataforma de computação sem servidor orientada a eventos que oferece a capacidade de executar seu código na nuvem sem se preocupar com a infraestrutura subjacente.

### O Cloud Functions oferece as seguintes vantagens fundamentais:

* Na maioria dos casos, quando você deseja executar qualquer tipo de código corporativo para funções operacionais ou de computação, é necessário ter uma máquina virtual hospedando e executando seu código. O Cloud Functions evita o esforço e a complexidade de manter suas próprias máquinas virtuais.
* Hospedar sua própria infraestrutura pode ser caro em relação ao benefício, especialmente se você precisar executar o código apenas algumas vezes por dia. O Cloud Functions, por outro lado, é efêmero, aumentando e diminuindo sob demanda, maximizando assim a eficiência e a economia à medida que você aproveita os recursos faturáveis. Você paga apenas pelo tempo que seu código leva para ser executado.
Relacionado ao item anterior, você pode configurar o Cloud Functions para disparar em resposta a eventos no ambiente, reduzindo ou eliminando a necessidade de ativação manual
* Minha parte favorita é que agora você pode escrever o código em Python 3 (beta) e, claro, JavaScript (Node.JS).

### Workflow
![estrutura-solucao-api-bigquery](https://github.com/ManoelCabral1/Prints/blob/main/estrutura-solucao-api-bigquery.png)

1. Cloud Scheduler, um serviço de tarefas cron gerenciado que permite que qualquer aplicativo chame operações de lote, Big Data e infraestrutura em nuvem. O Cloud Scheduler executa uma tarefa ao enviar uma solicitação HTTP ou uma mensagem Cloud Pub/Sub para um destino específico em uma programação recorrente. O gerenciador do destino executa a tarefa e retorna uma resposta. Se a tarefa for realizada, um código de sucesso (2xx para HTTP/AppEngine e 0 para Pub/Sub) será retornado para o Cloud Scheduler. Se a tarefa falhar, um erro será enviado para o Cloud Scheduler, que tentará novamente até que o número máximo de tentativas seja atingido. O Cloud Scheduler inicia o processo com um job programado para rodar uma vez ao dia que dispara uma messagem para o Cloud Pub/Sub.

2. O Pub/Sub é um serviço de mensagens em tempo real totalmente gerenciado que permite o envio e o recebimento de mensagens entre aplicativos independentes. 

3. A mensagem e repassada a Cloud function que em resposta executa o código de conexão e coleta dos dados da API Rest, em seguida persiste os dados no Bigquery

### Código da cloud function (python 3)

*bigquery_conector.py*

```
from google.cloud import bigquery
from google.cloud.bigquery import Table


def set_client():

    client = bigquery.Client()
    
    return client

def list_datasets(client):
    datasets = list(client.list_datasets()) 
    project = client.project
    if datasets:
        print("Datasets in project {}:".format(project))
        for dataset in datasets:
            print("\t{}".format(dataset.dataset_id))
    else:
        print("{} project does not contain any datasets.".format(project))

def create_dataset(client, dataset_id):
    dataset_id = client.project + '.' + dataset_id
    dataset_ref = bigquery.DatasetReference.from_string(dataset_id, default_project=client.project)
    dataset = bigquery.Dataset(dataset_ref)
    dataset.location = "US"
    dataset = client.create_dataset(dataset)
    print("Created dataset {}.{}".format(client.project, dataset.dataset_id))

def create_table(client, schema, table_name):
    table = client.project + '.' + 'dados.' + table_name
    table_id = Table.from_string(table)

    table = bigquery.Table(table_id, schema=schema)
    table = client.create_table(table)
    print(
        "Created table {}.{}.{}".format(table.project, table.dataset_id, table.table_id)
    )


def insert_rows(client, rows_to_insert):
    table = client.project + '.' + 'dados.dados_api'
    table_id = Table.from_string(table)

    errors = client.insert_rows_json(table_id, rows_to_insert) 
    if errors == []:
        print("New rows have been added.")
    else:
        print("Encountered errors while inserting rows: {}".format(errors))
```

*api_conector.py*

```
import requests
from requests.exceptions import RequestException
from datetime import date, timedelta

def set_dates():
    ''' função para setar o intervalo de data do endpoint da api, no caso d-1.'''

    delta1 = timedelta(days=2)
    delta2 = timedelta(days=1)

    data1 = date.today() - delta1
    data2 = date.today() - delta2

    return {
         "datainicial": data1.strftime("%Y-%m-%d"),
         "datafinal": data2.strftime("%Y-%m-%d")
   }



def format_url(id):
    ''' função para formatar o url do endpoint'''

    dates = set_dates()

    dates_str = f"&datainicial={dates['datainicial']}&datafinal={dates['datafinal']}"

    url = "http://api.xxx.com.br/indicadores/indicadoresEmpresa?chave=6ba108b1097b48238d414b9296cb05efles43" + 
          "&empresas[]=" + id + dates_str + "&periodo=1"

    return url

def extract_rows(row):
     '''função para extrair os campos de interesse no arquivo JSON'''

     keys_to_extract = ['IdEmpresa', 'NomeEmpresa', 'Data', 'FB_Fas', 'FB_Engajamento', 'FB_Reacoes', 
                       'FB_Comentarios', 'FB_Compartilhamentos', 'FB_Interacoes', 
                       'FB_Diferenca_Seguidores', 'TW_Seguidores', 'TW_Curtidas', 'TW_Retweets', 
                       'TW_Engajamento', 'TW_Interacoes', 'TW_Diferenca_Seguidores', 
                       'IN_Seguidores', 'IN_Curtidas', 'IN_Comentarios', 'IN_Engajamento', 
                       'IN_Interacoes', 'IN_Diferenca_Seguidores']

     return {key: row[key] for key in keys_to_extract}


def get_data():
    ''' função para conexão e coleta dos dados da api, faz a requisão dos dados de uma lista de empresas, uma de cada vez'''

    IDs = ["3528", "5395", "7226", "8376", "8378", "9008", "9601", "11165", "11797", "13890"]
    
    data = []

    for id in IDs:
        url = format_url(id)
        try:
            res = requests.get(url)

        except RequestException as e:
            continue
        try:
           res = res.json()[1]
        except KeyError as e:
            continue

        data.append(res)

    
    subset = []
    for row in data:
        subset.append(extract_rows(row))

    return subset
```
A api em questão não tem uma resposta para erro bad request (400) ou not found (404), no caso de alguma das empresas (IDs) não for encontrada. Mas se a requisição for bem sucedida o método res.json() retorna uma lista com dois elementos, e se falhar uma lista com um só elemento, dessa forma ocorre um KeyError (erro de índice de lista), então o código volta ao topo do loop eliminando os dados com erro.

*main.py*

```
from bigquery import*

from api_conector import get_data

def hello_pubsub(event, context):
    """Triggered from a message on a Cloud Pub/Sub topic.
    Args:
         event (dict): Event payload.
         context (google.cloud.functions.Context): Metadata for the event.
    """
    client = set_client()
    rows = get_data()
    insert_rows(client, rows)
```

A função hello_pubsub é ativada por evento, no caso a mensagem do pub/sub, assim que que o evento é disparado a função inicia o processo de coleta e persistência dos dados.

## Conclusão
As configurações em todas as ferramentas usadas devem ser feitas no console da Google Cloud Plataform. Lá existem vários tutoriais mostrando como usa-lás.

Lembrando que a carga diária faz o número de linhas da tabela crescer com o tempo, e o armazenamento de dados na nuvem tem custos, mesmo que baixos. Então dependendo do caso é aconselhável implementar métodos de backup dos registros mais antigos em formatos mais baratos, como csv no Cloud Storage (O data lake da GCP), não apliquei backup neste projeto porque ele é de curto prazo.
