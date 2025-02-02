Perfeito! Vamos agora detalhar ao máximo a Fase 2 do projeto "LlamaFlow", que é a fase de **Desenvolvimento do Workflow n8n**. Nesta fase, vamos criar o fluxo que coletará dados, prepará-los, enviá-los para o Nextflow e monitorar o progresso.

**Fase 2: Desenvolvimento do Workflow n8n (Detalhado)**

**Objetivo:** Criar um workflow no n8n que automatize a coleta de dados de treinamento, sua transformação para o formato adequado e o disparo do pipeline do Nextflow através de uma requisição HTTP. Além disso, vamos incluir um passo para monitorar a execução do Nextflow (opcional)

**Atividade 1: Criar um Webhook no n8n (Detalhado)**

*   **1.1 Abrir o n8n no Navegador:**
    *   **Ação:** Acessar `http://localhost:5678` no navegador (ou o endereço onde você instalou o n8n).
    *   **Ação:** Fazer login (se necessário).
*   **1.2 Criar um Novo Workflow:**
    *   **Ação:** Clicar em "Workflows" no menu lateral.
    *   **Ação:** Clicar em "Add Workflow" (ou similar).
    *   **Ação:** Dar um nome ao workflow (ex: "LlamaFlow Data Ingest").
*   **1.3 Adicionar um Nó "Webhook":**
    *   **Ação:** Clicar no botão "+" para adicionar um nó.
    *   **Ação:** Buscar por "Webhook" e selecionar o nó "Webhook".
    *   **Ação:** Configurar o nó:
        *   **Method:** Escolher "POST" (para receber dados enviados).
        *   **Path:** Escolher um caminho para a URL do webhook (ex: `/data-ingest`).
        *   **Note:** A URL completa do webhook (ex: `http://localhost:5678/webhook/data-ingest`) será mostrada no nó. Copie essa URL para usar posteriormente.
*   **1.4 Salvar o Workflow:**
    *   **Ação:** Clicar em "Save" para salvar o workflow.
*   **1.5 Ativar o Workflow:**
    *   **Ação:** Clicar no toggle na parte superior do workflow para ativar o workflow.

**Atividade 2: Adicionar Nó de Manipulação de Dados (JSON) (Detalhado)**

*   **2.1 Adicionar o Nó "Function":**
    *   **Ação:** Clicar no botão "+" após o nó "Webhook".
    *   **Ação:** Buscar por "Function" e selecionar o nó "Function".
*   **2.2 Configurar o Nó "Function":**
    *   **Ação:** Expandir a seção "Code" (ou similar).
    *   **Ação:** Escrever um script JavaScript para processar os dados recebidos do webhook.
        *   **Exemplo de Script:**
            ```javascript
            const data = $input.all(); // Recebe todos os dados de entrada
            
            if (!data || data.length === 0) {
                return []; // Retorna um array vazio se não houver dados
            }
          
            const transformedData = data.map(item => {
                const payload = item.json; // Assume que os dados estão no formato JSON
            
                if(payload && typeof payload === 'object') {
                 //Transformação dos dados
                 const transformed = {
                    text: payload.text.trim(),
                    source: payload.source || 'unknow'
                    };
                 
                 return { json: transformed};
                }
               
                return { json: payload }; // Retorna dados originais caso não sejam objeto json
            });
             
            return transformedData;
            ```
       * **Explicação:**
            *   O script recebe os dados do nó anterior (o nó Webhook) como input.
            *   Faz um tratamento dos dados, neste caso o script espera um array de objetos, pega o campo `text` e remove espaços desnecessários.
            *   Retorna o payload transformado ou original.
*   **2.3 Salvar o Workflow:**
     *   **Ação:** Clicar em "Save" para salvar o workflow.

**Atividade 3: Adicionar Nó S3/MinIO (Armazenamento) (Detalhado)**

*   **3.1 Adicionar o Nó "S3":**
    *   **Ação:** Clicar no botão "+" após o nó "Function".
    *   **Ação:** Buscar por "S3" e selecionar o nó "Amazon S3".
*   **3.2 Configurar o Nó "S3":**
    *   **Ação:** Selecionar "Credentials" e configurar as credenciais do S3 ou MinIO:
        *   **Access Key ID:** Preencher com o valor do `MINIO_ROOT_USER`.
        *   **Secret Access Key:** Preencher com o valor do `MINIO_ROOT_PASSWORD`.
        *   **Region:** Preencher com a região do seu S3 (ex: `us-east-1`).
        *   **Endpoint:** Preencher com `http://localhost:9000` para MinIO ou com o endereço do seu S3.
        *   **Force Path Style:** Marcar essa opção, se estiver usando o MinIO.
    *   **Ação:** Selecionar "Operation" e escolher "Upload".
        *   **Bucket Name:** Preencher com o nome do bucket que você criou (ex: `llamaflow-data`).
        *   **File Name:** Escolha um nome de arquivo para cada upload (pode ser usando a função expression como: `data-{{$now}}.json`).
        *   **Data to Upload:** Selecionar "Binary" e em "Binary Property" adicionar `data`
*   **3.3 Salvar o Workflow:**
     *   **Ação:** Clicar em "Save" para salvar o workflow.

**Atividade 4: Adicionar Nó HTTP Request (Nextflow) (Detalhado)**

*   **4.1 Adicionar o Nó "HTTP Request":**
    *   **Ação:** Clicar no botão "+" após o nó "Amazon S3".
    *   **Ação:** Buscar por "HTTP Request" e selecionar o nó "HTTP Request".
*   **4.2 Configurar o Nó "HTTP Request":**
    *   **Ação:** Preencher a "URL" da API do Nextflow Tower (ex: `http://<nextflow-tower-ip>:8080/api/workflows/launch`) ou o seu servidor que irá executar o pipeline.
        * Se estiver executando localmente, este campo poderá ficar vazio, caso esteja controlando por linha de comando.
    *   **Ação:** Definir o "Method" para "POST"
    *   **Ação:** Configurar a aba "Body" para incluir as informações de acesso ao seu repositório nextflow e parâmetros para executar o pipeline:
        * **Exemplo para Nextflow Tower:**
        ```json
            {
              "repository": "https://github.com/<seu-usuario>/<seu-repo>.git",
              "revision": "main",
              "params": {
                "input": "{{$json[\"fileName\"]}}"
              },
            "computeEnvId": 2
            }
        ```
           * `repository`: URL do seu repositório no GitHub.
           *  `revision`: branch do seu repositório nextflow (normalmente `main`).
           *  `params`: parametros do seu pipeline, neste caso passamos um arquivo que será o nome do arquivo salvo no S3.
           * `computeEnvId`: Id do ambiente de computação no Nextflow Tower.
       * **Exemplo para executar na linha de comando:**
        *  Deixe o campo `URL` em branco.
        *  Defina o `Method` para `GET`.
        *  Na aba "Query Parameters" adicione um parâmetro:
          * `command`: com o valor `nextflow run <caminho do seu workflow.nf> -params.input='{{"http://localhost:9000/llamaflow-data/" + $json["fileName"]}}'`
        * Nesta configuração o n8n irá enviar uma requisição `GET` com o comando para execução do pipeline.
    *   **Ação:** Configurar a aba "Authentication" (se necessário).
        *   Se estiver usando Nextflow Tower, usar o tipo "Header Auth" com a key `Authorization` e o valor `Bearer <seu-token>`.
*   **4.3 Salvar o Workflow:**
     *   **Ação:** Clicar em "Save" para salvar o workflow.

**Atividade 5: Adicionar Nó de Espera/Monitoramento (Opcional) (Detalhado)**

*   **5.1 Adicionar o Nó "Wait":**
    *   **Ação:** Clicar no botão "+" após o nó "HTTP Request".
    *   **Ação:** Buscar por "Wait" e selecionar o nó "Wait".
    *   **Ação:** Configurar o nó "Wait" para pausar o workflow por alguns segundos (ex: 5 segundos).
*   **5.2 Adicionar o Nó "HTTP Request" (Monitoramento):**
    *   **Ação:** Clicar no botão "+" após o nó "Wait".
    *   **Ação:** Buscar por "HTTP Request" e selecionar o nó "HTTP Request".
    *   **Ação:** Preencher a "URL" da API do Nextflow Tower para obter o status da execução do workflow (ex: `http://<nextflow-tower-ip>:8080/api/runs/<workflowId>`).
        * Se estiver usando a linha de comando para executar, poderá usar o status do comando anterior.
    *   **Ação:** Definir o "Method" para "GET" (para obter o status).
    *   **Ação:** Configurar a aba "Authentication" se necessário.
* **5.3 Adicionar Nó de tratamento de erros**
    * **Ação:** Clicar no botão "+" após o nó "HTTP Request" (Monitoramento).
    * **Ação:** Buscar e selecionar o nó "IF".
    * **Ação:** Configurar o nó "IF", na aba `Condition` use:
      * "Left Value" `{{ $json.status }}`
      * "Comparsion" `is not equal to`
      * "Right Value" `SUCCESS`
    * **Ação:** Adicionar após o nó "IF" o nó "Send Email" para notificar caso o pipeline falhe.
      * Configure o nó `Send Email` com seu email e o tipo de notificação.
*   **5.4 Salvar o Workflow:**
     *   **Ação:** Clicar em "Save" para salvar o workflow.

**Atividade 6: Testar o Workflow n8n (Detalhado)**

*   **6.1 Preparar Dados de Teste:**
    *   **Ação:** Criar um arquivo JSON de teste (ex: `test.json`).
        ```json
        [
            {
                "text": "     This is a test text.    ",
                "source": "test_data"
            },
          {
                "text": " This is another test text.  ",
                 "source": "test_data2"
           }
        ]
        ```
*   **6.2 Enviar Dados de Teste:**
    *   **Ação:** Usar um cliente HTTP (como Postman ou curl).
    *   **Ação (Curl):** Abrir um terminal ou prompt de comando e digitar o seguinte:
        ```bash
        curl -X POST -H "Content-Type: application/json" -d @test.json http://localhost:5678/webhook/data-ingest
        ```
        *   **Observação:** Ajustar a URL do webhook com a que você copiou no passo 1.3.
    *   **Ação (Postman):** Configurar o Postman para enviar uma requisição POST para o webhook, incluindo o corpo JSON.
*   **6.3 Verificar o Resultado:**
    *   **Ação:** Abrir o n8n e acessar a execução do workflow.
    *   **Ação:** Verificar se os dados foram processados corretamente (nó "Function").
    *   **Ação:** Verificar se o arquivo foi armazenado no S3 (nó "Amazon S3").
    *   **Ação:** Verificar se a requisição HTTP para o Nextflow Tower (ou para a linha de comando) foi enviada (nó "HTTP Request").
    *   **Ação:** Verificar se o status da execução foi verificado (nó de monitoramento).

**Resultado Final da Fase 2:**

Ao final da Fase 2, você terá:

*   Um workflow no n8n que recebe dados via webhook.
*   Um nó para transformar os dados para o formato adequado (JSON).
*   Um nó que envia os dados transformados para o bucket no S3/MinIO.
*   Um nó que dispara o pipeline do Nextflow através de uma requisição HTTP.
*   (Opcional) Um nó de espera que verifica a finalização do Nextflow.

Com isso, você estará pronto para iniciar o desenvolvimento do pipeline do Nextflow. Lembre-se de testar cada nó e configuração para garantir que tudo funcione como esperado.

Se tiver alguma dúvida, fique à vontade para perguntar. 😊
